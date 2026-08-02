# Verify the SDK install

Don't claim "done" until you've confirmed content actually renders.

## 1. The SDK loaded and initialized

In the running app's browser (DevTools console, or a headless browser if you have
one wired):

- `window.usertour` exists and has `init` / `identify` / `start` functions.
- No Usertour-related errors in the console.
- The network tab shows the SDK loader request succeeding.

## 2. A user is identified

- Confirm `usertour.identify(...)` runs after login with your canonical user id
  (see identify.md). Log the id you pass and confirm it matches an `externalId`
  that exists in Usertour (look the user up via the MCP `get_user` /
  `list_users`, or upsert a test user with that exact id).

## 3. Published content appears

- Make sure there is **published** content whose targeting includes your test
  user (a flow with start-rules that match, or call `usertour.start(contentId)`
  to force one for a deterministic check).
- Load the page where it should appear and confirm the flow / checklist / launcher
  renders. `usertour.start("<contentId>")` is the most reliable smoke test — it
  bypasses start-rule conditions and proves the SDK + theme + content pipeline
  works end to end.
- **Asserting via DOM/automation (not eyeballing)?** The SDK renders surfaces into
  a same-origin `<iframe class="usertour-widget-surface-viewport">` under a
  `#usertour-widget` host — so a top-level `querySelector` / text-wait finds
  nothing even when content IS showing. Read the iframe's `contentDocument` (or
  just assert the host + iframe exist) instead of concluding "nothing rendered."
  The SDK swaps the iframe surface per step, so re-query it each step — a
  `contentDocument` reference held from a previous step goes stale (reads null),
  and a stale null can look like a trigger that didn't fire. And don't assume
  YOURS is the only widget: other published content in the environment (an old
  flow, a resource-center launcher, an announcement) may auto-start for your
  test user at the same time and stack on screen — expected, not a broken
  install. Enumerate every `iframe.usertour-widget-surface-viewport` and pick
  yours by its step text; never grab "the first iframe".
- **Driving (not just reading) the widget programmatically** has two extra
  gotchas: accessibility snapshots show the iframe as an opaque "Content Frame"
  (no clickable uids inside — dispatch events via `contentDocument` instead),
  and some controls (e.g. the resource-center launcher) ignore a bare
  `.click()` — dispatch the full pointer sequence
  (`pointerdown → mousedown → pointerup → mouseup → click`) when a programmatic
  click seems to do nothing. When matching list rows / buttons by text, compare
  `textContent.trim() === label` EXACTLY — a `startsWith`/contains match can hit
  the same words inside a greeting or description elsewhere in the panel.
  (Ready-made read/click/rating snippets: the `usertour-content-authoring`
  skill → patterns.md → "Driving it in a real browser".)
- **Never "clean up" with `usertour.endAll()`.** Banner, launcher, and resource
  center are SINGLE-SESSION — one session per user for life — and `endAll()`
  ends them permanently for that test user (they simply never appear again; no
  error). To re-test one, delete its session instead:
  `list_sessions({ contentId, userId })` → `delete_session(id)` via the MCP.

## 3½. Data-plane check that works under automation (local / self-hosted)

"Watch the websocket in the network panel" is **not executable** in most
automated setups: the SDK's websocket typically does NOT appear in
DevTools-protocol request lists or `performance.getEntriesByType('resource')`,
and on a local/self-hosted install the request list is dominated by Cloud CDN
entries (`js.usertour.io` bundle/CSS) — which is the EXPECTED look in the
local-dev shape (the bundle stays on Cloud; see self-hosted.md), not a
misconfiguration. Assert **server-side** instead:

1. the render assertion above passes, then
2. MCP `get_user(<the id you identified>)` — a `first_seen_at` stamped at
   page-load time proves `identify()` reached YOUR instance;
3. MCP `list_sessions` filtered to your content + user — a session recorded for
   content that only exists on your instance is sufficient proof the SDK's data
   plane (`WS_URI`) points at it.

One render + one MCP read closes the loop on both the identify linkage and the
endpoint pointing — strictly stronger than eyeballing a network panel.

## 4. Security check

- Grep the app source for `utp_` — the API token must **not** be present in any
  client-side code. Only the environment token belongs in `init()`.

## If nothing shows

Work down this list:
1. `window.usertour` missing → loader snippet didn't run / wrong place.
2. SDK present but no content → user not identified, or `identify()` id ≠ the
   `externalId` the content targets (most common — see identify.md).
3. Right user, still nothing → content not published, or its start-rules don't
   match this user/page. Test with `usertour.start(contentId)` to isolate.
4. Renders unstyled / broken → the content's version has no theme (an authoring
   issue, not an install issue).
