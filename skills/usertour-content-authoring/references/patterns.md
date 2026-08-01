# Design judgment (the craft)

The judgment the schema can't carry. Mechanics — step types, navigation by `key`,
`completeWhen`/`clickActions`, frequency, the markdown subset, `run_javascript`
being rejected, what each type needs to publish — live in `get_authoring_guide`;
don't transcribe them here. For exact fields use `get_content_schema`.

## Pick the right content type

| Want | Type |
|------|------|
| A guided multi-step walkthrough (steps must happen now, in order) | **flow** |
| A "get started" list of independent, self-paced tasks | **checklist** |
| A persistent help button / spotlight that opens a tooltip or runs an action | **launcher** |
| A one-time notice strip at the top/bottom of the page | **banner** |
| A help hub with tabs of content | **resource-center** |
| Release notes / product news users can revisit, with an unread badge or one-time popup | **announcement** |
| Fire an analytics event when conditions match, no UI | **tracker** |

Two opposite mistakes to avoid:
- Reaching for a linear **flow** by default — it forces a sequence and a "now",
  only right when steps truly are sequential. Several independent setup tasks
  belong in a **checklist** (resumable, user-paced).
- Padding a **checklist** with invented tasks when there's really one activation
  action — that belongs in a tight flow (2–3 steps) or a single launcher/tooltip.

Ongoing help belongs in a **launcher** / **resource-center**, not a tour.

**Banner vs announcement:** a banner is a one-shot notice (each user sees it
once, dismiss and it's gone forever — think maintenance windows); an
announcement is a FEED item (release notes, product news) users can revisit,
delivered with an unread badge or a one-time popup. Announcements reach users
ONLY through a resource center with an `announcement` block — publish alone
surfaces nothing (the guide's Announcements section has the wiring).

## Make the flow follow the user

A step that can only advance via its own button gets stranded the moment the user
acts on the page instead — the classic case is a tooltip on a nav link: the user
clicks the real link, the app navigates, and the flow is left behind on the old
step. Don't fix it by deleting the step — make it react: advance on the step's
target click, and/or when a condition becomes true (reached a page, an element
appeared) no matter how the user got there, and keep a button as the fallback.
Layer all three so a contextual step feels smooth instead of "click-our-button"
linear. (The field shapes — `onClick`, `triggers` — are in `get_authoring_guide`.)

**For "the user completed an action," advance on the result — not the submit
click.** A submit / save / confirm click is only an *attempt*: if the action is
validation-gated, the click can fail and nothing happens, yet a step keyed on the
button being *clicked* advances anyway and falsely reports success. Key the advance
on the post-action change that proves it worked — the dialog/drawer closing
(`element` hidden), or a success element/toast appearing. (Clicks are fine for
actions that can't fail — opening a menu, navigating.) You usually only learn an
action is validation-gated by driving the app: do a partial/invalid submit and
watch whether the dialog actually closes.

## Sequence what auto-shows

If more than one piece can appear on its own (a welcome flow + a checklist + a
launcher), they pile onto the first screen at once. Show one, then reveal the next
when the first is done — gate the later surface's start rules on the first's state
(e.g. "the welcome flow has been seen and is now closed"). One focused thing at a
time. (How to gate on another content's state — `get_authoring_guide`.)

## One idea per surface

A single modal, tooltip, or checklist should carry one idea. The schema happily
lets you stack a dozen blocks, a dozen checklist items, or a paragraph of tooltip
copy — and an agent, unlike a designer watching it render, gets no signal that the
screen has become a wall. Showing too much at once is the most common way solid
content reads as noise: the user skims, misses the one action that mattered, and
dismisses it. Keep each step to a short headline + a sentence or two + one primary
action; keep a checklist to ~3–5 items. When there's genuinely more, split it
across flow steps or reveal it progressively — never all on the first screen.
(Companion to "Sequence what auto-shows": that's separate surfaces colliding;
this is one surface overloaded.)

## Segments are the targeting primitive

Attribute targeting goes through **segments**, not ad-hoc list filters: create a
condition `segment` (`create_segment`) and reference it with a
`{ "type": "segment", "segment": "<id>", "in": true }` condition. Segments are the
reusable, named unit — also how you stop re-showing content to users who already
converted (segment on the completion).

## Stable selectors

A target is only as durable as its selector, and the runtime targets the FIRST
element the selector matches. So make the selector resolve to exactly one element —
a stable `id` or `data-*` (`{ "selector": "[data-tour='create']" }`). If the
element has no stable selector, add one in the app source on that single element
(not a reused component or list row). If a stable selector still matches several
elements, pin the one you want with `nth` (0-based index of the match, range 0–4).
`text` is a separate refinement — it requires the matched element's exact visible
text, so it pins a specific content/state (e.g. `id` + `text`), but on its own it
does NOT search among matches (a non-unique selector with only `text` still hits
the first match, then fails its text check). Avoid `nth-child` / deep descendant
CSS that breaks on markup changes, and don't key on visible text alone (fragile to
copy changes and i18n). If you don't know the app's DOM, ask for a verified
selector rather than guessing — a flow that points at a missing or wrong element
renders nothing. (Exact `target` fields: `get_content_schema`.)

**Shared components need page scoping.** A reused component (a table toolbar, a
generic dialog) serving several pages emits the SAME selector everywhere — so
`[data-tour='toolbar-filter']` on the Tasks page also matches on Users, and the
runtime takes the first match on whatever page the step fires. Two patterns that
work: scope through a page-level ancestor
(`[data-page='tasks'] [data-tour='toolbar-filter']` — give each page container a
stable marker), or thread a `data-tour` PROP through the shared component so each
page instance renders its own value (`<DataTableToolbar dataTour="tasks-toolbar" />`).
Pair either with the step's `current_url` gating so the step can only fire on its
own page.

## Always verify — every interactive path, not just "does it appear"

`validate_content_version` proves it *can* publish, not that it *works* — and "it
rendered" isn't "it works" either. What slips past both are interaction bugs. Load
the app as the identified end-user and actually walk it:
- **Every checklist item** — click each: does it do what its row implies? (An item
  with no action, or only navigating while siblings launch a flow, feels like
  "nothing happened" even though it's valid.)
- **Every flow step** — the button path, the target/`onClick` path, AND the
  skip/dismiss path, on the page/state where the step's target actually exists.
- **Targets resolve** — a valid flow pointing at a missing selector renders nothing.
- **Gating** — the surface appears when (and only when) it should.

Don't claim "done" off the happy path alone.

### Driving it in a real browser (agents)

Usertour renders inside its OWN same-origin iframes, not the host DOM — so
`document.body.innerText` / a11y-tree queries never match what's on screen, and
an a11y snapshot shows only empty content frames. Read and drive the widget
through the iframe document instead:

```js
// read what's on screen (flow surface; banners use usertour-widget-banner-frame)
const frame = document.querySelector('iframe.usertour-widget-surface-viewport');
const text = frame?.contentDocument?.body?.innerText;

// click a widget button by its label
[...frame.contentDocument.querySelectorAll('button')]
  .find((b) => b.textContent?.trim() === 'Next')?.click();

// star ratings ignore a bare click — dispatch the full pointer sequence
const star = frame.contentDocument.querySelectorAll('[role="radio"], .star')[3];
for (const type of ['pointerdown', 'pointerup', 'click']) {
  star.dispatchEvent(new PointerEvent(type, { bubbles: true }));
}
```

Or just screenshot — pixels see through iframes. Two session traps while
testing: banner / launcher / resource center are SINGLE-SESSION (one per user,
for life) — to re-test, delete the old session (`list_sessions({ contentId,
userId })` → `delete_session`) rather than rotating externalIds; and never call
`usertour.endAll()` as "cleanup" — it permanently burns every single-session
surface for that user (recoverable only by that same session delete).
