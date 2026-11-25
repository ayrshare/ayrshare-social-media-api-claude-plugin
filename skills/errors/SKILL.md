---
name: errors
description: |
  Error explanation for the Ayrshare MCP server: translate an Ayrshare error code into a plain-English cause, a classification, and a fix. Use whenever a code or numeric error needs decoding: "what does Ayrshare error 156 mean", "why did that post fail with code 272", "explain this error", "what is error code 110 on Instagram", "how do I fix this Ayrshare error". Trigger when calling `mcp__ayrshare__explain_error`, and even without the word "Ayrshare": if the user surfaces a social-posting error code and wants to know what it means or how to fix it, this is the skill. CRITICAL: whenever ANY OTHER Ayrshare MCP tool returns a structured error code (for example `Error 156: ...`), look the code up here and explain or fix it rather than echoing the raw error back. For the shared auth model and the global retry-safety rule, see `../getting-started/SKILL.md`.
---

# Ayrshare MCP — Errors

One tool, the canonical way to decode any Ayrshare error code:

- `mcp__ayrshare__explain_error` — looks up an error `code` in the Ayrshare error catalog and returns a plain-English cause, a classification, and a short resolution hint.

It is the cross-cutting helper every other group skill points to: when a tool call fails with a structured error code (e.g. `Error 156: ...`), translate it here before surfacing anything to the user.

## Functions

| Tool | Purpose | Method + Endpoint | Required inputs | Optional inputs |
|------|---------|-------------------|-----------------|-----------------|
| `mcp__ayrshare__explain_error` | Explain an Ayrshare error code from the canonical error catalog | `GET /error-explainer` | `code` (string of digits, or a number) | `platform` (a platform identifier) |

The response gives three things:

- **Cause** — a plain-English description of what went wrong.
- **Classification** — one of `developer-fix` (your code/config/input is wrong), `user-action` (the end user or their linked account must act, e.g. re-auth, permissions, content rules), or `network` (a transient/platform-side issue).
- **Resolution hint** — a short suggested fix.

The classification is derived from the error's HTTP status, not from the meaning of the message: `401/402/403` → `user-action`, `5xx` → `network`, and every other `4xx` (including `400`) → `developer-fix`. So a relink-required error such as `156` ("…is not linked") or `272` (X/Twitter authorization issue) — both HTTP `400` — is labeled `developer-fix` even though the real fix is an end-user relink. Treat the label as a coarse routing hint and always read the `cause` and `resolution` text; a `developer-fix` can still carry an instruction meant for the end user.

## Auth

This tool is reached over the same authenticated connection as every other Ayrshare MCP tool. `explain_error` is account-level: it decodes an error code and does **not** take a `profileKey` argument (it is one of the few tools excluded from per-call profile selection). Its only inputs are `code` and `platform` (no `passthrough` object). See *Which tools accept `profileKey`* in `../getting-started/SKILL.md`.

## Usage guidance

- **This is the first stop after ANY failed tool call.** When `create_post`, `add_comment`, `get_platform_history`, or any other Ayrshare MCP tool returns a structured error code (e.g. `Error 156: ...`), call `explain_error` with that `code` (pass the failing `platform` if you have it) and surface the explanation. Do **not** echo the raw error or guess at the meaning.
- **Pass `platform` when you have it.** The same numeric code can mean different things per network; supplying the `platform` identifier (e.g. the platform that the failed call targeted) sharpens the cause and fix.
- **`code` accepts a string of digits or a number.** Pass the bare code from the error (e.g. `156`, `"272"`). Strip any surrounding text.
- **Act on the classification:**
  - `developer-fix` → usually the input/config/code is wrong; fix it before retrying (do not blindly re-call the failing tool). But because this bucket is just "a 4xx that isn't 401/402/403", it also catches end-user issues that happen to return `400` (e.g. an account relink, like codes `156`/`272`) — read the `resolution` and relay it to the end user when that is what it says.
  - `user-action` → the end user or their linked social account must do something (re-authenticate, grant a permission, change content); relay the resolution hint to them.
  - `network` → transient/platform-side; a single retry after a short delay is reasonable (matching the global 429/transient rule), but don't loop.
- **It does not fix the underlying call.** `explain_error` only decodes the code. After understanding it, address the root cause and re-run the original tool deliberately — don't auto-retry the failed write.

## Gotchas

- **No Ayrshare tools at all is not an error code.** If `explain_error` and every other Ayrshare tool are missing, ask the user to run `/mcp` first: if it shows the Ayrshare server as failed or disconnected, the problem is the connection, not the key, and they can reconnect it there. Otherwise the plugin's API key setting is most likely empty, so Claude Code did not start the server (or Claude Code is older than 2.1.119). This is the normal state right after updating from plugin version 0.2.2 or earlier, which read the key from an environment variable. Then tell the user to run `/ayrshare:setup`; do not ask for the key in the chat. See *Missing credentials* in `../getting-started/SKILL.md`.
- **Don't echo raw error codes at the user.** A bare `Error 272` is unhelpful. Always run it through `explain_error` and present the cause + fix.
- **The classification drives whether a retry is even sensible.** A `developer-fix` or `user-action` error will not resolve on retry — only `network`-class errors warrant the single transient retry. Never auto-retry a failed write on a 4xx just because you decoded it.
- **Provide `platform` for accuracy.** Omitting it still works but may give a more generic explanation when the code is platform-specific.
- **`explain_error` takes no `profileKey` argument.** It is account-level (one of the few tools excluded from per-call profile selection); the profile-scoped tools do accept a `profileKey` argument (see getting-started).
- **`explain_error` is itself a read.** If the lookup call fails (rare), surface it plainly; don't loop. (Mirrors the global retry-safety rule in `../getting-started/SKILL.md`.)
