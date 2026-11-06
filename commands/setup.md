---
description: Configure or rotate your Ayrshare API key (and optional Profile-Key / X BYOK) after installing the plugin
---

Help the user configure their Ayrshare credentials for Claude Code. Run this after installing the plugin, or to change a credential later.

The plugin asks for its credentials through Claude Code's plugin settings (`userConfig` in its `plugin.json`). Claude Code masks each value as it is typed and keeps it in the system's secure credential store, not in a settings file. The plugin's bundled MCP server reads them from there. You (Claude) never see, store, or write these values, so **never ask the user to paste a key into this chat**.

The four settings:

| Setting | Required | What it does |
|---|---|---|
| **Ayrshare API key** (`api_key`) | Yes | Sent as the `Authorization: Bearer` header on every call. From https://app.ayrshare.com, Settings, then API Key. |
| **Profile-Key** (`profile_key`) | No | Makes every call act as one client profile by default. Leave it empty to act on the account's primary profile. A `profileKey` tool argument on a single call still overrides it. |
| **X API Key** (`x_api_key`) and **X API Secret** (`x_api_secret`) | Only for X/Twitter (posting, analytics, messages) | Your X Developer App's OAuth 1.0a pair, sent as the `X-Twitter-OAuth1-Api-Key` and `X-Twitter-OAuth1-Api-Secret` headers. Set both or neither: with neither, any X call fails with `419`; with only one, it fails with `400`. Other networks are unaffected. |

**Surface note:** this configures **Claude Code** (CLI, IDE, and the desktop **Code** tab). Cowork does not prompt for plugin settings, so the plugin's tools do not load there yet; see the "Cowork and the desktop app" section in the README.

## Steps

1. **If the user pasted a key into the chat** (for example as an argument to this command): do not store or repeat it. Tell them it is now in this conversation, and that they should enter it in the settings dialog below instead. If they consider it exposed, they can generate a new key at https://app.ayrshare.com.

2. **Tell the user to open the plugin's settings dialog.** Claude cannot open it for them; they type it themselves:

   ```text
   /plugin configure ayrshare@ayrshare
   ```

   If they installed the plugin from a different marketplace name, the same dialog is under `/plugin`, on the **Installed** tab: select **ayrshare**, then **Configure options**.

   Then walk them through the fields:
   - **Ayrshare API key**: required.
   - **Profile-Key**: ask whether they want every call to default to one client profile. This is a neutral choice that depends on how they work; do not present either answer as recommended. If not, they leave it empty.
   - **X API Key** and **X API Secret**: ask "Do you use X/Twitter through Ayrshare (posting, analytics or messages)?" If yes, they enter both. If no, they leave both empty. Do not present skipping as recommended: anyone who makes any X call needs the pair.

   To change a value later, they open the same dialog again. Leaving a field as it is keeps its saved value.

3. **If they set up the plugin with version 0.2.2 or earlier**, those versions read the key from environment variables, and this version no longer does. Tell them their old `AYRSHARE_API_KEY`, `AYRSHARE_PROFILE_KEY`, `X_TWITTER_OAUTH1_API_KEY` and `X_TWITTER_OAUTH1_API_SECRET` entries now do nothing, and that they can delete them from wherever they put them: the `env` block of `~/.claude/settings.json`, `./.claude/settings.local.json` or `./.claude/settings.json`, or a shell profile. Let them do this themselves; do not open or read those files.

4. **Offer to clean up a stale duplicate server.** Older versions of this command created a separate `ayrshare` MCP server (via `claude mcp add`) with the key in a header. That extra server shadows or collides with the plugin and should be removed. Run `claude mcp list` and identify it carefully:
   - The **plugin's own** server is the one to keep. The plugin provides it from the bundled `.mcp.json`, and Claude Code lists it namespaced as `plugin:ayrshare:ayrshare` (URL `https://api.ayrshare.com/mcp`). **Never remove this one.**
   - A **stale duplicate** is a *separately added* server listed as plain `ayrshare` (no `plugin:` prefix) at `https://api.ayrshare.com/mcp`, i.e. an `ayrshare` entry that exists *in addition to* the plugin's, often flagged as "defined in multiple scopes" by `claude mcp list`. Only this one should be offered for removal, with `claude mcp remove ayrshare` (add `--scope user` or `--scope local` to match where it lives).
   - **Never remove an `ayrshare` server whose URL is the docs endpoint** (`https://www.ayrshare.com/docs/mcp`). That is the separate Ayrshare documentation MCP, not a duplicate; leave it untouched.
   - If the only `ayrshare`-related server is the plugin's own (`plugin:ayrshare:ayrshare`), there is nothing to clean up; do not prompt for removal.

5. **Tell the user how to finish and how to use it.**
   - After they save, Claude Code shows "Configuration saved." and reloads the plugin, which reconnects the server with the new values. If tools still return `401` or `403`, they should run `/reload-plugins` or restart Claude Code. If no Ayrshare tools appear at all, the API key setting is still empty, or their Claude Code is older than 2.1.119.
   - Then set expectations on invocation: the tools are used by **asking in plain English** (e.g. "show my recent Instagram posts", "post this to LinkedIn"), not by typing a slash command. `/ayrshare:setup` is the **only** slash command; the other tools fire on intent (the trigger-skills route them), so `/ayrshare:get_post_history` and similar will read as "unknown command."

## Notes

- One mechanism: the four settings above are exactly what the plugin's bundled `.mcp.json` reads (`${user_config.api_key}`, `${user_config.profile_key}`, `${user_config.x_api_key}`, `${user_config.x_api_secret}`). No duplicate servers and no settings files.
- The three optional settings default to empty. An empty header is treated by the server as not provided, so leaving one empty has no effect and causes no error.
- To set the values from a shell without a dialog (for example on a CI machine), on Claude Code 2.1.285 or later install the plugin, then pipe a JSON object of the values into `claude plugin configure ayrshare@ayrshare --values-stdin`, reading the key from the CI system's secret store. Older versions only take `--config api_key=...` on `claude plugin install`; a value on the command line can show up in shell history and in the process list, so use that only on a machine nobody else uses.
