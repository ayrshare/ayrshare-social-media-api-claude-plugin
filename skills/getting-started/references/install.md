# Ayrshare MCP Server: Installation

Full instructions for installing the Ayrshare MCP server in Claude Code. SKILL.md summarizes these; this file has the exact commands and the gotchas.

The Ayrshare MCP server is an **HTTP MCP server** at `https://api.ayrshare.com/mcp`. It authenticates with the header `Authorization: Bearer <API key>`, where the key is your **account-level Business plan API key** (from [app.ayrshare.com](https://app.ayrshare.com) → Settings → API Key). With the plugin, the key is the plugin's `api_key` setting; it is not a per-call argument.

## Option 1: Claude Code plugin (recommended)

Installs the MCP server plus the `/ayrshare:setup` command, the social-manager and profile-manager agents, and the trigger-based skills.

### Choose an install scope

First register this repo as a marketplace (one time):

```bash
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin
```

Then install the `ayrshare` plugin (`plugin@marketplace`) at the scope you want:

| Scope | Command | Where it's available |
|---|---|---|
| Global (default) | `claude plugin install ayrshare@ayrshare` | Every project on this machine |
| This project only | `claude plugin install ayrshare@ayrshare --scope local` | Current project, not committed to git |
| This project (shared) | `claude plugin install ayrshare@ayrshare --scope project` | Current project, committed to git with the team |

### Enter the key

After installing, open the plugin's settings dialog inside Claude Code:

```text
/plugin configure ayrshare@ayrshare
```

If the plugin came from a different marketplace name, the same dialog is under `/plugin`, on the **Installed** tab: select **ayrshare**, then **Configure options**. `claude plugin install` run from a shell does not show the dialog.

The dialog asks for the **Ayrshare API key** (required) and three optional settings (see *Optional configuration* below). Claude Code masks each value as it is typed and stores it in the system's secure credential store, not in a settings file. The plugin's bundled `.mcp.json` reads the values from there, so there is nothing else to set. `/ayrshare:setup` explains each field; it never asks for a key in the chat.

Saving reloads the plugin and reconnects its server. If a tool still returns 401/403, run `/reload-plugins` or restart Claude Code, then run `mcp__ayrshare__get_post_history` (a plan-agnostic read; `list_profiles` needs a Business plan) to confirm.

To rotate the key later, open the same dialog again.

Do not also run `claude mcp add`; a separate server is unnecessary and is the
common cause of a `403 / code 102` after setup. The plugin already provides its own
server (listed as `plugin:ayrshare:ayrshare`); a "duplicate" is an *extra*, separately
added `ayrshare` server at the same api URL (typically from a past `claude mcp add`,
shown as `ayrshare` defined in multiple scopes). `/ayrshare:setup` offers to remove only that
extra server, never the plugin's own and never the docs MCP.

### Upgrading from 0.2.2 or earlier

This version needs Claude Code 2.1.119 or later. Earlier plugin versions read the key from the `AYRSHARE_API_KEY` environment variable (and the optional `AYRSHARE_PROFILE_KEY`, `X_TWITTER_OAUTH1_API_KEY` and `X_TWITTER_OAUTH1_API_SECRET`). This version no longer reads them, and its tools do not load until you enter the key. Enter the same values once with `/plugin configure ayrshare@ayrshare`, then delete the old variables from the `env` block of your `settings.json` or your shell profile.

## Option 2: MCP only (no plugin)

If you only want the raw MCP tools without commands, agents, or skills:

```bash
claude mcp add ayrshare --transport http \
  https://api.ayrshare.com/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

Add `--scope local` to limit it to the current project. The key is stored permanently in Claude Code's config; run once and you're done.

## Option 3: Install without a dialog (CI / headless)

On Claude Code 2.1.285 or later, install the plugin, then pipe the values in as JSON on standard input, so the key never appears on a command line:

```bash
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin
claude plugin install ayrshare@ayrshare
printf '{"api_key":"%s"}' "$AYRSHARE_API_KEY" | claude plugin configure ayrshare@ayrshare --values-stdin
```

Here `$AYRSHARE_API_KEY` is a variable your CI system fills from its secret store; the plugin itself does not read it.

The keys are `api_key`, `profile_key`, `x_api_key` and `x_api_secret`; keys you leave out keep their saved values. Older Claude Code versions only take each setting as `--config KEY=VALUE` on `claude plugin install`. A value on the command line can show up in shell history and in the process list, so use `--config` only on a machine nobody else uses.

## Optional configuration

The plugin's bundled `.mcp.json` sends these optional headers from settings that default to
empty, so you turn each feature on by filling in the setting with
`/plugin configure ayrshare@ayrshare`. Leave a setting empty and its header is sent empty,
which the server treats as "not provided" (no effect, no error), so there is no manual
`.mcp.json` editing and nothing to undo on `claude plugin update`.

| Setting | Description |
|---|---|
| **Profile-Key** (`profile_key`) | Sets the `Profile-Key` header so the whole connection defaults to a specific client profile. For one-off or runtime profile selection, pass `profileKey` as a tool argument on a profile-scoped call instead; it takes precedence over the header and needs no restart. See the auth model in SKILL.md. |
| **X API Key** (`x_api_key`) + **X API Secret** (`x_api_secret`) | Your X Developer App's OAuth 1.0a key pair (API Key / Consumer Key and API Secret / Consumer Secret), required for any X/Twitter call (posting, analytics, messages) under the BYO-key mandate (effective March 31, 2026). These map to the `X-Twitter-OAuth1-Api-Key` / `X-Twitter-OAuth1-Api-Secret` headers, the only X BYO headers Ayrshare uses (one key pair per account, on every X-targeting request; no OAuth 2.0 client credentials or per-user access tokens). Values are never logged. Set **both** or neither: neither set → error `419` (`x_credentials_required`); only one set → error `400`. |

## Notes

- A **Business plan** key is required for the Profiles / multi-profile tools (`mcp__ayrshare__create_profile`, `mcp__ayrshare__list_profiles`, `mcp__ayrshare__generate_jwt_social_linking_url`). `generate_jwt_social_linking_url` mints a client's social-account linking URL for the target profile (set by the `profileKey` argument or the `Profile-Key` header; the argument wins) and additionally requires a provisioned social-linking domain (Business/Enterprise plans). Deleting profiles is done in the Ayrshare dashboard / REST API, not via an MCP tool.
- No account or key yet? Start a 28-day free trial of the Launch plan (see SKILL.md for the attributed signup link). Note the Launch trial does not include Profiles.
- After saving the settings, run `/reload-plugins` or restart Claude Code if a tool still returns 401/403.
