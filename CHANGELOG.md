# Changelog

## 0.2.3

- You now enter your Ayrshare API key in the plugin's own settings dialog, where Claude Code masks it and keeps it in your system's secure credential store. The optional Profile-Key and X API Key and Secret are settings in the same dialog.
- The plugin no longer reads `AYRSHARE_API_KEY`, `AYRSHARE_PROFILE_KEY`, `X_TWITTER_OAUTH1_API_KEY` or `X_TWITTER_OAUTH1_API_SECRET` from your environment.
- `/ayrshare:setup` now walks you through the settings dialog and no longer asks you to paste a key into the chat.
- The plugin now has an icon and links to the Ayrshare privacy policy, terms, documentation and support pages.

**Check this if** you set up the plugin with version 0.2.2 or earlier: after updating, run `/plugin configure ayrshare@ayrshare` in Claude Code and enter your API key (and any optional values you used). Until you do, the Ayrshare tools do not load. This version needs Claude Code 2.1.119 or later. You can then delete the old variables from your `settings.json` or shell profile.

To update, run `claude plugin marketplace update ayrshare`, then `claude plugin update ayrshare@ayrshare`, and restart Claude Code.

## 0.2.2

- The confirmation prompt now appears when the server comes from this plugin. Earlier versions did not show it. It now also covers `update_ad`, `boost_post`, `register_webhook`, and `unregister_webhook`.
- The `insights-analyst` agent now starts when the server comes from this plugin.

To update, run `claude plugin marketplace update ayrshare`, then `claude plugin update ayrshare@ayrshare`, and restart Claude Code.
