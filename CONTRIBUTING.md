# Contributing

## This is a public repository

Anything committed here is visible to the world the moment it is pushed. A leaked
credential is compromised instantly and **cannot be un-leaked** by deleting the commit
(push events and the API have already exposed it). Treat secret prevention as a
local, pre-push concern, not something CI can fix after the fact.

Never commit:

- API keys, bearer tokens, or `Authorization` header values with real keys
- Private keys (the Ayrshare account private key can mint linking URLs for every
  profile under the account)
- `.env` files (already covered by `.gitignore`)

Use placeholders in committed files: `${user_config.api_key}`, `Bearer KEY`, etc.

## Enable local secret scanning (one-time, per clone)

We use [TruffleHog](https://github.com/trufflesecurity/trufflehog) to catch secrets
before they are committed. After cloning:

```sh
brew install trufflehog          # macOS; other OSes: https://github.com/trufflesecurity/trufflehog#installation
git config core.hooksPath .githooks
chmod +x .githooks/pre-commit
```

The `pre-commit` hook scans your staged changes and blocks the commit if it finds
anything that looks like a secret, including findings it cannot verify as live (a
staged private key, for example). If TruffleHog is not installed, the hook warns and
lets the commit through, CI is the backstop in that case.

## CI backstop

Every pull request (and every push to `main`) runs the **TruffleHog Secret Scanning**
workflow (`.github/workflows/trufflehog.yml`). Any finding fails the check, including
unverifiable ones such as a private key. This check is **required** in branch
protection for `main`, so a red PR cannot be merged.

If a real placeholder ever trips the scanner (a fake-but-realistic example token in
docs), add a `trufflehog:ignore` comment on that line rather than weakening the scan.

If a secret ever does land on a branch: **rotate the key first** (it is already
exposed), then scrub the history.

## Authoring skills

Each skill's `SKILL.md` frontmatter `description` is validated more strictly by the
claude.ai Cowork org uploader and the community marketplace than by `claude plugin
validate` (which checks neither rule). To keep the plugin installable on every
surface:

- **Length:** keep each description **at most 1024 characters**, and aim well under
  it (target roughly 900 or fewer) so a small later edit does not breach the cap.
- **No angle brackets:** a description must not contain `<` or `>`; the validators
  read them as XML tags and reject the upload. Write `Error 156`, not `Error <code>`;
  write "under 24h", not "< 24h".

Enforcement: `scripts/check-skill-descriptions.py` checks both rules. It runs in the
`pre-commit` hook (once you set `core.hooksPath .githooks` above) and in the
**Validate Skills** CI workflow on every pull request and push to `main`. Run it
anytime with `python3 scripts/check-skill-descriptions.py`.
