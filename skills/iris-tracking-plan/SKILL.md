---
name: iris-tracking-plan
description: Use when the user wants to plan or document Mixpanel tracking for a Unifonic feature - finding which events/properties already exist, drafting new events or properties, or building/updating a journey (screens, triggers, events) in Iris. Also when placing journey triggers on screens from Figma designs.
---

# Iris tracking plan

Iris is Unifonic's source of truth for the Mixpanel tracking plan. You work through the `iris` MCP tools. Everything you create is a **draft** (journeys "Draft", events and properties "Proposed") that a person reviews in Iris before it ships, so draft confidently but never pretend something is final.

## If the Iris tools don't work

- **No `iris` tools at all** - the plugin can't reach Iris until `IRIS_URL` and `IRIS_TOKEN` are set in the user's shell and Claude Code is restarted. Point them to this plugin's README.
- **Connection refused / timeout** - Iris is only reachable on the Unifonic VPN. Ask the user to connect and retry.
- **401 / "invalid or revoked agent token"** - the token was revoked or mistyped. They create a new one in Iris under Settings → Agent tokens.

## Rules (Unifonic's Event Based Analytics Framework)

1. **Search before you create.** Run `search_events` before `create_event` and `search_properties` before `propose_property`, every time. Properties must come from the shared pool - propose a new one only when nothing existing fits, and say why the closest existing ones didn't.
2. **Never reuse an event name for a different purpose.** If a similar event exists but means something else, create a distinct, clearly named one and explain the difference.
3. **Naming.** Events are action-driven phrases in Title Case: "Account Created", "Campaign Form Submitted". Properties are snake_case: `campaign_id`, `failure_reason`.
4. **Every event carries `context`, `sub_context` and `action_type`** - Iris attaches them automatically. Set their real values on each trigger with `set_trigger_property_value`:
   - `context` - the distinct area of the platform or user journey (e.g. "Campaigns").
   - `sub_context` - the sub-level within it (e.g. "Campaign builder").
   - `action_type` - `user` if a person's action fires it, `system` if the platform does.
   Keep values consistent with the same product's other journeys (check one with `get_journey`); ask the user when unsure.
5. **Track what matters, not everything.** Tie events to the feature's goal or KPI. Prefer back-end state changes over front-end clicks ("Form Submitted" when the record is saved, not "Submit Button Clicked"). Never propose high-volume per-unit events (e.g. one per SMS sent).
6. **Warnings mean nothing was saved.** When a write returns `warnings`, read them. Reuse the existing item if it fits; only resubmit with `force: true` when yours is genuinely different.
7. **Always share the `url`** from tool results, and end with a summary of what you reused vs. created.

## Workflow A - Drafting tracking for a feature (PMs, e.g. via Baymax)

1. **Understand the feature**: its goal/KPI, the product, and the screens involved. Ask if any of these are unclear.
2. `list_product_lines` → pick the product (confirm with the user if ambiguous).
3. For each meaningful user action or outcome: `search_events` → reuse a fitting event, or `create_event` with a Title Case name, a plain-language `when_fired`, and `journey_id` once the journey exists.
4. For each piece of data the event needs: `search_properties` → reuse, or `propose_property` (snake_case, a real description, `enum_values` for ENUM). Then `attach_property_to_event`.
5. `create_journey`, then `create_journey_step` per screen, in order.
6. **Screens - Figma links first.** Ask whether the screens are in Figma. If so, ask for each screen's frame link (in Figma: right-click the frame → "Copy link to selection") and pass it as `figma_url` to `create_journey_step`, or call `import_figma_frame` for an existing step. Iris renders the frame itself - nobody needs a Figma seat. A link to the whole file (no `node-id`) won't work; ask for the frame link. If the result says Figma isn't connected, fall back to screenshots below.

   **Screenshots (no Figma)**: ask the user for each screen as an image **file path** (they can drag the file into the terminal). Upload it by running the `upload_screenshot_with` curl command that `create_journey_step` returned, with `<path-to-image>` replaced by the path. Run it exactly as given - it carries a one-hour signed link, so don't add headers or `$IRIS_TOKEN`. If the link expired, get a fresh one with `get_screenshot_upload_command`. Never base64 a screenshot yourself; pasted images can't be uploaded - you need the file.
7. `add_trigger` per interaction on its step, with `element_name` and `event_id`.
   - **Figma-imported step**: `list_figma_layers` (use `query`, e.g. "send") and pass the matching layer's id as `figma_node_id` - Iris places the marker exactly. Pick the element itself (usually an INSTANCE such as "Button / Primary"), not its label text. If no layer clearly matches, leave the marker off and list it as an open item - the PM can click the element on the step image in Iris to place it.
   - **Screenshot step**: give a rough normalized `box` only if you can clearly see where the element is, otherwise leave the marker off.
8. `set_trigger_property_value` for `context`, `sub_context`, `action_type` (and any value that's fixed for that trigger).
9. `create_journey_edge` from each trigger to the step it leads to.
10. Report: the journey link, events/properties reused vs. new, and open items (e.g. "no Figma layer matched 'Export CSV' - place its marker in Iris").

When the designs change later, `refresh_from_figma` re-renders a journey's Figma steps; markers follow their layers, and markers whose layer was deleted are reported back - tell the user which.

## Workflow B - Placing triggers from Figma by hand (fallback)

Only needed when Iris has no Figma connection (`import_figma_frame` says so) and the user has Figma access themselves - otherwise use Workflow A's Figma links. Use when the user gives you a journey (link or name - find it with `list_journeys`) and the Figma file for its screens.

1. `get_journey` to see its steps and triggers - often a PM drafted it already, with triggers that have no marker yet.
2. For each step, find the matching Figma frame and get it as an image file: **the whole frame, uncropped** (any scale works - markers are stored as fractions of the image). Either export the frame from Figma, or, if `FIGMA_TOKEN` is set, use the Figma REST API (`GET https://api.figma.com/v1/images/<file_key>?ids=<frame_id>&format=png`, then download the returned URL). Upload it by calling `get_screenshot_upload_command` for the step and running the command it returns with the file path filled in.
3. For each element that fires an event, get its box and the frame's box **in the same coordinate space** - `absoluteBoundingBox` for both is simplest (REST: `GET https://api.figma.com/v1/files/<file_key>/nodes?ids=<frame_id>,<node_id>`). If you only have the element's position relative to the frame, pass `frame_box` as `{x: 0, y: 0, width, height}`. Nested layers' positions are relative to their parent - add the offsets up to the frame.
4. Existing trigger → `update_trigger` with `figma`; new interaction → `add_trigger` with `figma` (and `event_id` if known). Always include `file_key` and `node_id`; set `element_name` to a clear name ("Send button"), not just the raw layer name. **Never invent a `test_id`** - set it only when the user gives the team's real one.
5. Check with `get_journey`: each marker's x/y should be between 0 and 1. A `warning` that the box extends outside the frame means the node/frame pair is wrong - fix it rather than leaving a clamped marker.

## Quick reference

| Need | Tool |
|---|---|
| Products | `list_product_lines` |
| Existing events / properties | `search_events`, `search_properties` |
| Journeys | `list_journeys`, `get_journey` |
| New event / property | `create_event`, `propose_property` |
| Event ↔ property | `attach_property_to_event` |
| Journey structure | `create_journey`, `create_journey_step`, `create_journey_edge` |
| Screen from Figma | `figma_url` on `create_journey_step`, `import_figma_frame`, `refresh_from_figma` |
| Screenshot file | run the curl command from `create_journey_step` / `get_screenshot_upload_command` |
| Triggers | `add_trigger`, `update_trigger` (`figma_node_id` from `list_figma_layers`), `connect_event_to_trigger` |
| Fixed values (`context`, `action_type`...) | `set_trigger_property_value` |
