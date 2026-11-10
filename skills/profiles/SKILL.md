---
name: profiles
description: |
  Profile management for the Ayrshare MCP server: create profiles, list profiles, and generate social-account linking URLs for client/customer profiles under a Business account. Use whenever someone wants to onboard a new client, set up a customer's social accounts, "add a profile", "create a workspace for a client", list existing profiles, recover a lost `profileKey`, or get a link a client opens to connect their own social networks. Trigger when calling `mcp__ayrshare__create_profile`, `mcp__ayrshare__list_profiles`, or `mcp__ayrshare__generate_jwt_social_linking_url`, and even when the user does not say "Ayrshare": agency or white-label social setup, managing multiple clients' accounts through an AI assistant, or "send my client a link to link their socials". `generate_jwt_social_linking_url` mints the single sign-on URL the client opens to connect their networks, targeting the sub-profile from the `profileKey` argument or `Profile-Key` header (argument wins).
---

# Ayrshare MCP — Profiles

Profile management is the **agency / multi-client layer** of Ayrshare. A *profile* is one client (or customer workspace) under your Business account; each owns its own set of connected social networks and its own `profileKey`. These three tools create profiles, list them, and mint the social-account linking URL a client opens to connect their networks. Together they cover client onboarding — see the onboarding sequence in `../getting-started/SKILL.md`.

## Functions

| Tool | Purpose | Method + Endpoint | Profile-scoped | Required inputs | Optional inputs |
|------|---------|-------------------|----------------|-----------------|-----------------|
| `mcp__ayrshare__create_profile` | Create a new client profile; returns its `profileKey` | `POST /profiles` | No (account-level) | `title` | `messagingActive`, `hideTopHeader`, `hideLogo`, `topHeader`, `subHeader`, `disableSocial`, `team` (requires `email`), `email`, `tags` |
| `mcp__ayrshare__list_profiles` | List all profiles under the Business account and their linked platforms | `GET /profiles` | No (account-level) | — | `title`, `refId`, `hasActiveSocialAccounts`, `includesActiveSocialAccounts`, `isByokLinked`, `actionLog`, `limit`, `cursor`, `include` |
| `mcp__ayrshare__generate_jwt_social_linking_url` | Mint a single sign-on social-account **linking URL** for one sub-profile, the link a client opens to connect their own networks | `POST /profiles/generateJWT` | **Yes**, targets the sub-profile set by the `profileKey` argument or `Profile-Key` header (required; argument wins) | `profileKey` (or set it via the `Profile-Key` header) | `logout`, `redirect`, `allowedSocial`, `verify`, `expiresIn` (Max Pack), `email` (object, `to` required, optional `bcc`/`termsUrl`/`privacyUrl`/`company`/`contactEmail`; Max Pack) |

Full input schemas and example payloads/responses are in [`references/schemas.md`](references/schemas.md).

> `mcp__ayrshare__generate_jwt_social_linking_url` mints the single sign-on linking URL a client opens to connect their social accounts; it does **not** run the OAuth flow itself (the client completes that in their browser) and it does **not** link anything on its own. There is still **no MCP tool to delete a profile**. See *Linking a client's social accounts* below.

## Auth

`create_profile` and `list_profiles` are **account-level**: they authenticate with the **Business API key** only: the `Authorization: Bearer` header the MCP server sends, from the plugin's `api_key` setting (`/plugin configure ayrshare@ayrshare`). You never supply a `profileKey` as the *auth* key, and neither of these two takes a `profileKey` argument (both ignore the `Profile-Key` header too; they are account-level). (`generate_jwt_social_linking_url`, the third tool here, is **profile-scoped** instead: its target sub-profile comes from the `profileKey` argument or the `Profile-Key` header, the argument winning. See below.)

**How profile scoping actually works.** `create_profile` and `list_profiles` take no `profileKey` and **ignore the `Profile-Key` header**; they are account-level. To *act as* a specific client profile (post, fetch analytics, pull history, or mint that client's linking URL), supply that client's `profileKey` either as a **tool argument** on the call (per call; it wins when both are set) or as the **`Profile-Key` connection header** in the MCP client config (the `headers` block of `.mcp.json`; the default for every call on the connection). With neither set, calls act on the account's primary/Business profile. The `profileKey` that `create_profile` returns is the value you pass as the argument or place in that header. The profile-scoped tools, including the linking tool covered next, accept it either way. Full model: `../getting-started/SKILL.md`.

**`generate_jwt_social_linking_url` is profile-scoped.** It mints the linking URL for the target sub-profile, which is **required** for this tool: supply it as the `profileKey` argument or the connection's `Profile-Key` header (the argument wins), exactly as you would to post or pull analytics as that client. You do **not** supply any signing credentials: the MCP server derives the account's private key and onboarding domain **server-side** from your authenticated account, so no `X-Ayrshare-*` headers are needed. It does require the account to have a **provisioned social-linking domain (Business/Enterprise plans)**; without one it returns a clear "no social-linking domain provisioned" error. Optional BYOK X headers (`X-Twitter-OAuth1-Api-Key` / `X-Twitter-OAuth1-Api-Secret`) still apply if you link X with your own developer app.

`create_profile` / `list_profiles` require a **Business plan**. A Launch/free-trial key will fail them even though it is otherwise valid.

## Linking a client's social accounts

Creating a profile does **not** link any social networks to it — a fresh profile has zero connected platforms. To connect a client's accounts you generate a **single sign-on linking URL** and the client opens it to OAuth their own networks.

- **`mcp__ayrshare__generate_jwt_social_linking_url` (in-MCP).** Pass the client's `profileKey` as the `profileKey` argument (or set the connection's `Profile-Key` header) to get back a hosted `url` (valid 5 minutes by default, or `expiresIn` on the Max Pack). Hand that URL to the client; they open it in a browser and link their accounts. No private key or domain needed; the server derives them (requires a provisioned social-linking domain, Business/Enterprise). It only *mints* the link; the OAuth happens in the client's browser, not via any MCP tool.
- **Ayrshare dashboard** (Social Accounts → link), or the **REST API** (`POST /profiles/generateJWT`) directly — equivalent alternatives to `generate_jwt_social_linking_url` if you are not driving it through the MCP.

After the client's accounts are linked, act as that profile by passing its `profileKey` argument (or setting the connection's `Profile-Key` header) and verify with the History tools (`mcp__ayrshare__get_post_history` / `mcp__ayrshare__get_platform_history`).

## Usage guidance

- **Onboarding a client** runs in order: `create_profile` (account-level) → capture the returned `profileKey` → `generate_jwt_social_linking_url` to mint the linking URL for that profile (pass the `profileKey` as the argument, no header change or restart needed) → hand the URL to the client to OAuth their accounts → act as that profile (via the `profileKey` argument or `Profile-Key` header) and verify with the History tools. See the full sequence in `../getting-started/SKILL.md`.
- **Capture the `profileKey`** from `create_profile` immediately. It is sensitive, shown once (the API cannot return it again), and is what you pass as the `profileKey` argument (or place in the `Profile-Key` header) to operate as that client. Treat it like a credential.
- **Lost a `profileKey`?** It cannot be recovered through the API — `list_profiles` does **not** return profile keys (the `GET /profiles` call omits them for security). Retrieve a lost key from the **Ayrshare dashboard**; don't create a duplicate profile just because a key was misplaced.
- **`team` profiles** require an `email`: set `team: true` only together with the team member's `email`, or the call fails.
- **`disableSocial`** takes an array of network names to disable for that profile; `topHeader`/`subHeader`/`hideTopHeader`/`hideLogo` control the white-label linking-page chrome.

## Gotchas

- **Scoping is the `profileKey` argument or the `Profile-Key` header.** `create_profile` / `list_profiles` are account-level (Business key) and **ignore** both. `generate_jwt_social_linking_url` is profile-scoped: it mints the link for the target sub-profile, required via the `profileKey` argument or `Profile-Key` header (the argument wins). No tool ever uses a `profileKey` as the *auth* key.
- **`generate_jwt_social_linking_url` mints the link; it does not link accounts itself.** A new profile starts empty. It returns the URL a client opens to OAuth their networks — the actual linking happens in the client's browser, not via any MCP tool. (The dashboard and REST `POST /profiles/generateJWT` are equivalent alternatives.) It needs **no** signing headers (the server derives the private key and domain) but requires a provisioned social-linking domain (Business/Enterprise).
- **No delete tool.** The MCP cannot delete or offboard a profile. If a user asks to remove a profile, tell them it must be done in the Ayrshare dashboard or via the REST API — do not claim an MCP tool can do it.
- **`team` without `email` fails.** If `team: true`, `email` is required.
- **`profileKey` is sensitive.** `create_profile` returns it once and the API never returns it again. Capture it, store it like a secret, and never echo it back in plain logs. A lost key is recoverable only from the Ayrshare dashboard.
- **`list_profiles` returns `title`/`refId`/linked platforms — not `profileKey`.** The `GET /profiles` call omits keys for security, so `list_profiles` cannot recover a lost key (use the dashboard for that). Use it to find a profile or its `refId`, not as a reason to re-create a profile.
- **Requires a Business plan.** Profiles and multi-client management are Business-only. A valid Launch/trial key still fails here.
- **Never auto-retry `create_profile` on a 4xx.** Call `mcp__ayrshare__explain_error`, surface it, and stop. (Mirrors the global retry-safety rule in `../getting-started/SKILL.md`; 429 gets at most one retry.)
