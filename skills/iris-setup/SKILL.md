---
name: iris-setup
description: Use when a PM or designer needs to connect their Claude Code to Iris for the first time, or reconnect after getting a new or revoked agent token. Sets up the `iris` MCP server by writing directly to Claude Code's own settings (not a shell profile), then verifies it works and walks through troubleshooting if it doesn't.
---

# Iris setup

You're doing this setup yourself, with your own file tools - don't ask the user to run shell commands for it. That avoids the two most common failure modes: a multi-line terminal command getting mangled by copy-paste, and a shell-profile env var that the desktop app (or anything else not launched from that shell) never sees.

## 1. Get the one thing only the user can give you

Ask for:
- **Iris URL** - the company's Iris host (e.g. `https://<iris-host>`). If they don't know it, tell them to ask whoever rolled out Iris.
- **An agent token** - created in Iris under **Settings → Agent tokens → New token**, shown once. If they don't have one yet, walk them to that page first, then have them paste the full token string here.

Never ask the user to type or paste the token into a terminal, a file, or anywhere outside this chat - writing it to disk is your job, not theirs. Once you've saved it (step 2), tell them not to paste it again; no need to repeat a secret that's already stored.

## 2. Write it to Claude Code's own settings, not a shell profile

Read `~/.claude/settings.json` if it exists - it may already hold permissions, hooks, or other config, so merge into it rather than overwriting the file. Create it if it doesn't exist.

Merge in a top-level `env` object:

```json
{
  "env": {
    "IRIS_URL": "<what the user gave you>",
    "IRIS_TOKEN": "<what the user gave you>"
  }
}
```

If `env` already has other keys, keep them - add `IRIS_URL`/`IRIS_TOKEN` alongside, don't replace the object. Validate the JSON parses before writing it.

**Why settings.json and not `export` in `~/.zshrc`:** a shell export only reaches Claude Code when it's launched from a shell that sourced that profile. The desktop app, or anything launched from Finder/Spotlight/Dock, won't see it - that's a real macOS behavior, not a one-off bug. `settings.json` is read by Claude Code itself, CLI or desktop, so there's one source of truth instead of two to keep in sync.

## 3. Get the plugin installed and the session restarted

If the `iris` MCP tools aren't already available in this session, have the user run:
```
/plugin marketplace add erikdandrade/iris_tools
/plugin install iris@unifonic-iris
```
(skip this if their agent's repo already enables it via `.claude/settings.json`'s `extraKnownMarketplaces`/`enabledPlugins` - see the main README's "For agent maintainers" section.)

Either way, the env vars you just wrote only take effect in a fresh process - have them fully quit and reopen Claude Code (`Cmd+Q`, not just closing the window, matters for the desktop app) or start a new CLI session.

## 4. Verify it yourself - don't make the user guess

Once restarted, call `list_product_lines` yourself. Report the actual result:
- **Works** - tell them what came back, e.g. "Connected - Iris has N product lines: ...". Setup's done.
- **No `iris` tools available at all** - the plugin isn't installed, or Claude Code wasn't actually restarted. Confirm both before anything else.
- **Connection refused / timeout** - not on the company VPN. Have them connect and retry.
- **401 "invalid or revoked agent token"** - the token was mistyped or already revoked. Have them create a new one in Iris and go back to step 1.

## Quick reference

| Symptom | Cause | Fix |
|---|---|---|
| No `iris` tools | plugin not installed, or session not restarted | `/plugin install iris@unifonic-iris`, then fully restart |
| Connection refused / timeout | off the Unifonic VPN | connect, retry |
| 401 invalid/revoked token | token mistyped or revoked | new token in Iris, redo steps 1-2 |
| Desktop app still can't connect after the settings.json edit | app wasn't fully quit, just the window was closed | quit with `Cmd+Q`, then reopen |
