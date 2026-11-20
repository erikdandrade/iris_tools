# Iris plugin for Claude Code

Lets Claude Code - on its own, or as part of an agent like **Baymax** - work with **Iris**, Unifonic's source of truth for the Mixpanel tracking plan. PMs and designers use it the same way: there's no separate "designer mode" that only places triggers, everyone can draft the same full tracking plan.

- find which events and properties already exist (fuzzy search catches near-duplicates)
- draft new events and properties following Unifonic's [Event Based Analytics Framework](https://unifonic.atlassian.net/wiki/spaces/UPP/pages/1985806477/Event+Based+Analytics+Framework+-+Mixpanel)
- build journeys: screens, the triggers on them, and the events they fire
- import screens from Figma frame links and place triggers precisely on their layers

Everything an agent creates lands in Iris as a **draft** attributed to you, for someone to review.

## What's inside

| File | What it does |
|---|---|
| `.mcp.json` | Connects to Iris's MCP endpoint (`$IRIS_URL/api/agent/mcp`) with your personal token |
| `skills/iris-tracking-plan/SKILL.md` | The rules and workflows the agent follows (search before creating, naming, `context`/`sub_context`/`action_type`, Figma placement) |
| `skills/iris-setup/SKILL.md` | Walks your agent through connecting itself to Iris - no shell profile editing |
| `.claude-plugin/` | Plugin and marketplace manifests |

## Setup (each PM / designer, once)

1. **Be on the Unifonic VPN** - Iris is only reachable there. (Claude Code itself works off-VPN; only the Iris tools won't connect.)
2. **Install the plugin** - skip this if your agent's repo (e.g. Baymax) already enables it, see below. In Claude Code:

   ```
   /plugin marketplace add erikdandrade/iris_tools
   /plugin install iris@unifonic-iris
   ```

3. **Get a token.** In Iris, open **Settings → Agent tokens → New token**, name it after the agent (e.g. "Baymax" or your own name), and copy the two lines it shows.
4. **Tell your agent to set up Iris for you** and give it the URL and token when it asks. The `iris-setup` skill writes them into Claude Code's own settings itself, has you restart, and verifies the connection - no editor, no shell profile.

Check it worked by asking: *"What events does Iris have for campaigns?"*

If you don't maintain an agent repo like Baymax, you're done - skip the "For agent maintainers" section below, it's only about wiring the plugin into another agent's own repo.

No Figma seat needed: give the agent a Figma frame link and Iris renders the screen and places markers on its layers itself. (Precise trigger placement from your own Figma access - handy for designers, but open to anyone - needs Figma's MCP or a `FIGMA_TOKEN` on a Dev Mode seat; see the skill for how.)

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
| No `iris` tools in Claude Code | `IRIS_URL` / `IRIS_TOKEN` aren't set, or Claude Code hasn't restarted since - ask your agent to run the `iris-setup` skill |
| Connection refused / timeout | Connect to the VPN |
| 401 "invalid or revoked agent token" | Create a new token in Iris (Settings → Agent tokens) |
| Screenshot won't upload | Give the agent the image as a file path; pasted images can't be uploaded. Claude Code asks before running the `curl` upload command - approve it |
| "Figma isn't connected in Iris" | Iris's admin hasn't connected Figma yet, or the token lapsed (Settings → Integrations in Iris). Use screenshot files meanwhile |
| Figma import takes minutes or times out | Figma's API sometimes stalls a request for 2-4 minutes. Wait and retry; a fix (timeouts + background import) is planned in Iris |
| Tool not found: `import_figma_frame`, `list_figma_layers`... | The Iris server is older than this plugin - see Versions below |

## Versions

| Plugin | Needs Iris | Changes |
|---|---|---|
| 0.4.2 | same as 0.4.1 | Added a reconciliation step to the workflow: `add_trigger` can leave a trigger with no event (expected mid-Figma-walk), but the agent must now either connect it, or call it out explicitly as an open item, before reporting - never leave one silently unresolved |
| 0.4.1 | same as 0.4.0 | Doc fix: Iris's Figma connection now lives in Settings → Integrations (admin-managed, DB-backed), not a `FIGMA_ACCESS_TOKEN` env var - corrected the Troubleshooting row |
| 0.4.0 | same as 0.3.0 | Merged the old "PM" and "designer fallback" workflows into one - drafting events/properties/journeys is the same workflow for anyone; your own Figma access is a precision upgrade for trigger placement, not a separate, more limited role |
| 0.3.0 | same as 0.2.0 | Added the `iris-setup` skill - your agent configures `IRIS_URL`/`IRIS_TOKEN` in Claude Code's own settings itself, instead of a shell profile; setup steps above reflect this |
| 0.2.0 | Figma import (Iris commit `10e8844`, 2026-11-02) with `FIGMA_ACCESS_TOKEN` set on the server | Screens come from Figma frame links (`figma_url`, `import_figma_frame`); markers go on Figma layers (`list_figma_layers` + `figma_node_id`); `refresh_from_figma`. The designer-by-hand workflow is now the fallback |
| 0.1.0 | Agent tokens + MCP server | First release: search/draft events and properties, build journeys, upload screenshots |
