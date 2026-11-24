---
name: iris-tracking-plan
description: Use when the user wants to plan or document Mixpanel tracking for a Unifonic feature - finding which events/properties already exist, drafting new events or properties, or building/updating a journey (screens, triggers, events) in Iris. Also when placing journey triggers on screens from Figma designs.
---

# Iris tracking plan

Iris is Unifonic's source of truth for the Mixpanel tracking plan. You work through the `iris` MCP tools. Everything you create is a **draft** (journeys "Draft", events and properties "Proposed") that a person reviews in Iris before it ships, so draft confidently but never pretend something is final.

## If the Iris tools don't work

- **No `iris` tools at all** - `IRIS_URL`/`IRIS_TOKEN` aren't set or Claude Code hasn't been restarted since. Use the `iris-setup` skill to fix it - don't ask the user to edit a shell profile by hand.
- **Connection refused / timeout** - Iris is only reachable on the Unifonic VPN. Ask the user to connect and retry.
- **401 / "invalid or revoked agent token"** - the token was revoked or mistyped. They create a new one in Iris under Settings → Agent tokens.

## Rules (Unifonic's Event Based Analytics Framework)

1. **Search before you create.** Run `search_events` before `create_event` and `search_properties` before `propose_property`, every time. Properties must come from the shared pool - propose a new one only when nothing existing fits, and say why the closest existing ones didn't.
2. **Pick the property's scope deliberately, and don't propose "just in case."** `EVENT` scope for data specific to that action/moment (e.g. `plan_type` chosen on this submit); `USER` scope for an attribute of the person/account used for cohorting that persists across events (e.g. `subscription_status`, `signup_date`) - search both scopes before proposing. Only add a property a known question or segment actually needs; it's easier to add one later than to prune one nobody uses.
3. **Never reuse an event name for a different purpose.** If a similar event exists but means something else, create a distinct, clearly named one and explain the difference.
4. **Naming.** Events are action-driven phrases in Title Case: "Account Created", "Campaign Form Submitted". Properties are snake_case: `campaign_id`, `failure_reason`.
5. **Every event carries `context`, `sub_context` and `action_type`** - Iris attaches them automatically. Set their real values on each trigger with `set_trigger_property_value`:
   - `context` - the distinct area of the platform or user journey (e.g. "Campaigns").
   - `sub_context` - the sub-level within it (e.g. "Campaign builder").
   - `action_type` - `user` if a person's action fires it, `system` if the platform does.
   Keep values consistent with the same product's other journeys (check one with `get_journey`); ask the user when unsure. **If `set_trigger_property_value` fails with "property isn't attached"**, that's a known gap on some Iris instances (the three properties were never created server-side) - don't let it block the rest of the journey; skip it and name it as an explicit open item in your report instead of retrying or guessing a workaround.
6. **Track what matters, not everything.** Tie each event to the feature's goal or metric by working backward from it: what's the first action in the flow, the last, the milestones between, and what would signal a drop-off or dead end - that drop-off signal is often the single most valuable event to capture. Prefer back-end state changes over front-end clicks ("Form Submitted" when the record is saved, not "Submit Button Clicked"). Never propose high-volume per-unit events (e.g. one per SMS sent).
7. **Warnings mean nothing was saved.** When a write returns `warnings`, read them. Reuse the existing item if it fits; only resubmit with `force: true` when yours is genuinely different.
8. **Always share the `url`** from tool results, and end with a summary of what you reused vs. created.

## Workflow - Drafting tracking for a feature

This is the same workflow for anyone - a PM working from a ticket, a designer working from their own Figma file, whoever. Nothing here is gated by role; the only thing that differs is where your screens and trigger precision come from.

**Research and propose first - never build before the user signs off.** Phase 1 is read-only research that ends in a written proposal (events, properties, journey shape, and the reasoning behind them); Phase 2 is the actual build in Iris, and it only starts once the user has seen the proposal and explicitly said to go ahead. Even for a request that sounds like "just set up tracking for X," default to proposing first - don't skip straight to creating things because the ask seemed simple.

### Phase 1 - Research and propose (no writes to Iris)

1. **Understand the feature**: its goal/KPI, the product, and the screens involved. Then map the flow before picking events - what's the first action, the last, the milestones between, and what would signal a drop-off or dead end (Rule 6); that's your event list. Ask if any of this is unclear.
2. **Figure out the product.** `list_product_lines` (read-only) and pick the one that fits; flag to the user if more than one could, or if none does - there's no tool to create a product, so one would need to be added on Iris's **Products** page before you continue. Don't treat this as a blocker to finishing the proposal; name it as an open item instead.
3. **Check what already exists.** For each meaningful action or outcome, `search_events` - and `search_properties` for the data each would need - to see what's reusable vs. what would be new. Do **not** call `create_event`, `propose_property`, `attach_property_to_event`, or any other write yet; this step is only to inform the proposal.
4. **Work out the journey shape.** Figure out the screens/overlays (`search_pages`, `search_overlays`) and, if screens are in Figma, which frames and element layers correspond to each interaction (`list_figma_layers`, `list_figma_section_frames` - all read-only). Don't call `create_journey`, `create_journey_step`, or `add_trigger` yet.
5. **Write up the proposal** and share it with the user before doing anything else:
   - The product line, and any open item from step 2.
   - The event list: for each, whether it's reused or new, its Title Case name, a plain-language `when_fired`, and why it's the right one for the flow (tie back to Rule 6 - goal/KPI, milestone, or drop-off signal; back-end state change vs. a raw click).
   - The properties each event needs: reused vs. new, scope (`EVENT` vs `USER`, per Rule 2) and why.
   - The journey/screen flow: steps in order, and which are Pages vs. Overlays.
   - Open items or judgment calls you made (e.g. ambiguous product line, no Figma access yet, an event you considered but left out and why).
6. **Ask explicitly**: does the user want you to implement this in Iris? If they want changes, revise the proposal and ask again - don't build a partially-approved version. Only move to Phase 2 on a clear yes.

### Phase 2 - Build in Iris (after approval)

7. For each event in the approved proposal: `create_event` with a Title Case name, a plain-language `when_fired`, and `journey_id` once the journey exists (reused events need no action here). For each property: `propose_property` (snake_case, a real description, `enum_values` for ENUM) then `attach_property_to_event` (reused properties just get attached).
8. `create_journey`, then for each screen: `search_pages` first - reuse a fitting Page, or pass a new `page_name` to `create_journey_step` to create one (warnings mean a near-duplicate was found; resubmit `force: true` only if genuinely different). `create_journey_step` per screen, in order.
   - **Not every step is a Page.** If an interaction opens a modal, dropdown, or menu rather than navigating to a new destination, that's an **Overlay**, not a Page - it always belongs to the Page it appears on, never a standalone thing. `search_overlays` (scoped to that page) first, then pass `overlay_name` alongside `page_name` to `create_journey_step` (or `update_journey_step` for a step that already has its page). Same warning/`force` behavior as pages.
9. **Screens - Figma links first.** Ask whether the screens are in Figma. If so, ask for each screen's frame link (in Figma: right-click the frame → "Copy link to selection") and pass it as `figma_url` to `create_journey_step`, or call `import_figma_frame` for an existing step. Iris renders the frame itself - nobody needs a Figma seat. A link to the whole file (no `node-id`) won't work; ask for the frame link. If the result says Figma isn't connected, fall back to screenshots below.

   **Screens grouped in a Figma Section**: if several of the flow's screens are grouped under one Section in Figma, ask for the section's link instead (right-click the section itself → "Copy link to selection") and call `list_figma_section_frames` once rather than asking for one frame link per screen. Use each returned `figma_url` exactly as if it had been pasted directly - only the frames inside that section, never sibling frames or sections elsewhere in the file. A nested section's own children aren't unwrapped further; ask for that nested section's link separately if needed.

   **Screenshots (no Figma)**: ask the user for each screen as an image **file path** (they can drag the file into the terminal). Upload it by running the `upload_screenshot_with` curl command that `create_journey_step` returned, with `<path-to-image>` replaced by the path. Run it exactly as given - it carries a one-hour signed link, so don't add headers or `$IRIS_TOKEN`. If the link expired, get a fresh one with `get_screenshot_upload_command`. Never base64 a screenshot yourself; pasted images can't be uploaded - you need the file.
10. `add_trigger` per interaction on its step, with `element_name` and `event_id`.
    - **Default, Figma-imported step**: `list_figma_layers` (use `query`, e.g. "send") and pass the matching layer's id as `figma_node_id` - Iris places the marker exactly. Pick the element itself (usually an INSTANCE such as "Button / Primary"), not its label text. If no layer clearly matches, leave the marker off and list it as an open item - anyone can click the element on the step image in Iris to place it later.
    - **Screenshot step**: give a rough normalized `box` only if you can clearly see where the element is, otherwise leave the marker off.
    - **If you have your own Figma access** (e.g. a Figma MCP connection): this is a precision upgrade over `list_figma_layers`, not a separate role or a fallback gated on "Iris has no Figma connection" - use it any time you have Figma access and want exact geometry. Get the frame as an image, uncropped (export it, or if `FIGMA_TOKEN` is set, `GET https://api.figma.com/v1/images/<file_key>?ids=<frame_id>&format=png` then download the returned URL); for a step with no image yet, upload it via `get_screenshot_upload_command`. For the element, get its box and the frame's box **in the same coordinate space** - `absoluteBoundingBox` for both is simplest (`GET https://api.figma.com/v1/files/<file_key>/nodes?ids=<frame_id>,<node_id>`); if you only have a position relative to the frame, pass `frame_box` as `{x: 0, y: 0, width, height}`, adding up nested layers' offsets to the frame. Pass `figma` (`file_key`, `node_id`) on `add_trigger`/`update_trigger` - existing trigger with no marker yet → `update_trigger`, new interaction → `add_trigger` (with `event_id` if known). **Never invent a `test_id`** - set it only when you know the team's real one. If the response says the element isn't visible in the step's image (scrolled off-screen or clipped), no marker was placed - don't try to force one; it usually means the element is genuinely outside what this step's image shows, not a wrong node/frame pair. Leave it as an open item, same as an unmatched layer. If the step was imported a while ago, try `refresh_from_figma` first - a newer import may include content that used to get cropped out. A warning that the box extends only slightly outside the frame still gets a marker (clipped to what's visible) and is usually fine as-is.
11. `set_trigger_property_value` for `context`, `sub_context`, `action_type` (and any value that's fixed for that trigger).
12. `create_journey_edge` from each trigger to the step it leads to.
13. **Reconcile triggers before reporting.** `add_trigger` lets you place a marker with no `event_id` yet (e.g. while walking Figma layers before you've decided what fires on that element) - that's expected mid-workflow, but never leave it unresolved silently. Check `get_journey`: for every trigger you created, either it has an event, or you call `connect_event_to_trigger` once you know it, or it's explicitly named as an open item in your report. A trigger with no event and no mention in the report is the one outcome to avoid.
14. Report: the journey link, events/properties reused vs. new, and open items (e.g. "no Figma layer matched 'Export CSV' - place its marker in Iris", "trigger on 'Submit' has no event yet - confirm which one fires it before it ships"). Once the plan is actually finalized (not after every intermediate step), also share the journey's `share_url` - the no-login link meant for pasting into a ticket - and ask whether they'd like it added to the ticket.

When the designs change later, `refresh_from_figma` re-renders a journey's Figma steps; markers follow their layers, and markers whose layer was deleted are reported back - tell the user which.

## Quick reference

| Need | Tool |
|---|---|
| Products | `list_product_lines` |
| Existing events / properties | `search_events`, `search_properties` |
| Pages / Overlays | `search_pages`, `search_overlays` (scoped to one page) |
| Journeys | `list_journeys`, `get_journey` |
| New event / property | `create_event`, `propose_property` |
| Event ↔ property | `attach_property_to_event` |
| Journey structure | `create_journey`, `create_journey_step` (`page_name`/`overlay_name`), `update_journey_step`, `create_journey_edge` |
| Screen from Figma | `figma_url` on `create_journey_step`, `import_figma_frame`, `refresh_from_figma` |
| Screens grouped in a Figma Section | `list_figma_section_frames` |
| Screenshot file | run the curl command from `create_journey_step` / `get_screenshot_upload_command` |
| Triggers | `add_trigger`, `update_trigger` (`figma_node_id` from `list_figma_layers`), `connect_event_to_trigger` |
| Fixed values (`context`, `action_type`...) | `set_trigger_property_value` |
