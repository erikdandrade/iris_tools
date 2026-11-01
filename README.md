# Iris plugin for Claude Code

Lets Claude Code (and agents built on it, like **Baymax**) work with **Iris**, Unifonic's source of truth for the Mixpanel tracking plan:

- find which events and properties already exist (fuzzy search catches near-duplicates)
- draft new events and properties following Unifonic's [Event Based Analytics Framework](https://unifonic.atlassian.net/wiki/spaces/UPP/pages/1985806477/Event+Based+Analytics+Framework+-+Mixpanel)
- build journeys: screens, the triggers on them, and the events they fire
- place triggers precisely from Figma designs (Design team)

Everything an agent creates lands in Iris as a **draft** attributed to you, for someone to review.

## What's inside

| File | What it does |
|---|---|
| `.mcp.json` | Connects to Iris's MCP endpoint (`$IRIS_URL/api/agent/mcp`) with your personal token |
| `skills/iris-tracking-plan/SKILL.md` | The rules and workflows the agent follows (search before creating, naming, `context`/`sub_context`/`action_type`, Figma placement) |
| `.claude-plugin/` | Plugin and marketplace manifests |

## Setup (each PM / designer, once)

1. **Get a token.** In Iris, open **Settings → Agent tokens → New token**, name it after the agent (e.g. "Baymax"), and copy the two lines it shows.
2. **Add them to your shell profile** (`~/.zshrc` on a Mac), then open a new terminal:

   ```bash
   export IRIS_URL="https://<iris-host>"
   export IRIS_TOKEN="iris_..."
   ```

3. **Be on the Unifonic VPN** when using it - Iris is only reachable there. (Claude Code itself works off-VPN; only the Iris tools won't connect.)
4. **Install the plugin** - skip this if your agent's repo (e.g. Baymax) already enables it, see below. In Claude Code:

   ```
   /plugin marketplace add erikdandrade/iris_tools
   /plugin install iris@unifonic-iris
   ```

Check it works by asking: *"What events does Iris have for campaigns?"*

Designers using the Figma workflow also need Figma's MCP (or a `FIGMA_TOKEN` for the Figma REST API) on a seat with Dev Mode access.

## For agent maintainers (e.g. Baymax)

Add this to your agent repo's `.claude/settings.json`. Anyone who clones the repo and trusts it in Claude Code is prompted to install the Iris plugin - nothing else to wire up:

```json
{
  "extraKnownMarketplaces": {
    "unifonic-iris": {
      "source": { "source": "github", "repo": "erikdandrade/iris_tools" }
    }
  },
  "enabledPlugins": {
    "iris@unifonic-iris": true
  }
}
```

Each user still does steps 1-3 above (their own token, so their work is attributed to them and can be revoked individually).

Alternatively, add this repo as a git submodule and start Claude Code with `--plugin-dir path/to/iris_tools`.

The plugin's rules live in this repo, so improvements to them reach every agent on its next plugin update - no changes needed in the agent's own repo.

## Troubleshooting

| Symptom | Fix |
|---|---|
| No `iris` tools in Claude Code | `IRIS_URL` / `IRIS_TOKEN` not set in the shell Claude Code was started from - set them and restart |
| Connection refused / timeout | Connect to the VPN |
| 401 "invalid or revoked agent token" | Create a new token in Iris (Settings → Agent tokens) |
| Screenshot won't upload | Give the agent the image as a file path; pasted images can't be uploaded. Claude Code asks before running the `curl` upload command - approve it |
