---
name: usertour-content-authoring
description: Author and publish Usertour in-app onboarding content — flows (tours), checklists, launchers, banners, announcements (release notes / product news), surveys, trackers — via the Usertour API/MCP. Use when the user mentions Usertour, building a product tour / onboarding / in-app guide / checklist, or asks to create, update, validate, translate / localize, or publish Usertour content. Biases toward retrieving the live schema and authoring guide from the Usertour MCP over pre-trained knowledge.
---

# Usertour content authoring

Your baked-in knowledge of Usertour's content shapes may be incomplete or
outdated. **Prefer retrieval from the Usertour MCP over pre-training** for the
representation, field schemas, and conventions — they are self-describing and
always current.

## Retrieval sources (consult these first)

| Source | How | Use for |
|--------|-----|---------|
| `get_authoring_guide` (MCP tool) | no-arg call at the start → core + a table of contents; then fetch the sections for your content type in ONE array call (`section: [...]`) | Lifecycle + per-type publish requirements come inline; step types, conditions, start rules, markdown subset etc. are named sections |
| `get_content_schema` (MCP tool) | `get_content_schema({ type })` | The exact write body — `steps` for `flow`, `data` for checklist/launcher/banner/tracker/announcement/resource-center. **Fetch before authoring a non-flow type** (its `data` is polymorphic). |
| `validate_content_version` (MCP tool) | dry-run a draft | The publishable-or-not check + precise errors |
| v2 docs | https://docs.usertour.io (API reference v2 → Concepts) | Human reference: blocks, conditions & actions, rules, type data |

If the `usertour` MCP is not connected, tell the user to configure it (see the
plugin's `.mcp.json` / README) — do not guess field shapes.

## Design before you author

Onboarding is a means to an activation goal, not a tour for its own sake. Before
calling any tool, decide (infer from context — don't interrogate):
- **Who** — the segment / lifecycle stage.
- **One action** — the single behavior that means "activated"; build toward it.
- **When** — the trigger moment (first login, reaching a page, an empty state).
- **Done** — what signals success (a click, an event, a checklist completion).

Then pick the form and right-size it. Defaults that make content *good*, not just
valid (override only with a reason):
- **One activation action → a tight flow (2–3 steps) or a single
  launcher/tooltip.** Don't pad it into a checklist by inventing tasks.
- **Several genuinely independent setup tasks → checklist** (resumable). Don't
  force them into a linear flow.
- **First-run flow = 3–5 steps**, one idea and one primary action each. More →
  split into a checklist or several context-triggered flows.
- **Trigger on context, not on load** — a tooltip when the user reaches its
  page/element; a welcome modal `frequency: once`; help on an empty state. Never
  fire on every page or every session.
- **Always escapable** — a skip/dismiss; don't block the UI unless necessary;
  don't re-show completed or dismissed content.
- **One audience per piece** — split by segment, don't make one flow serve all.

Avoid the default bad onboarding you'll otherwise reach for: the 8-step grand
tour that fires on load, narrates obvious UI ("This is the menu"), and forces a
sequence on unrelated tasks.

## Workflow

Design the experience first (above), then build it:

1. **Read the guide** — call `get_authoring_guide` with no args (the core +
   a table of contents), then fetch the sections for your content type in one
   array call — pick by each section's `appliesTo`, or take the `guideSections`
   list that `get_content_schema` returns for the type.
2. **Pick a theme** — `list_themes`; pass a `themeId` to `create_content` (every
   visual type needs one). Use the `isDefault` theme if unsure.
3. **Fetch the schema** — `get_content_schema({ type })` for the body you'll write.
4. **Create** — `create_content({ type, name, themeId })` → returns `editedVersionId`.
5. **Author** — `update_content_version({ contentId, versionId, steps | data, startRules })`.
   Top-level fields merge, but a list you send (`steps`, checklist `items`) replaces
   that whole list — see `get_authoring_guide`.
6. **Validate** — `validate_content_version`; fix every error before publishing.
7. **Translate, if the project is multilingual** — `list_localizations`; anything
   beyond the default locale means users may be reading a translation. (Asked for a
   language that isn't there? `create_localization` adds it — copy the tag and
   language name from the catalog in the guide's `localization` section rather
   than improvising: `name` is what machine translation is asked to translate
   INTO. Locale tools need the localization scopes — without them, the same codes
   come back from the expand below.) After ANY text change, read
   `get_content_version` with
   `expand: ["localizations"]` and bring every enabled locale back to zero
   `missing` / `outdated`:
   `get_version_localization` for the units, then ONE `update_version_localization`
   call per locale. Nothing blocks a publish with stale translations — skip this
   and those users keep the old wording. Asked to translate existing content? Same
   loop; a published version must be forked first (`create_content_version`
   carries the translations along). Rules: the guide's `localization` section.
8. **Publish** — `publish_content` (per environment; idempotent).
9. **Verify** — load the app as the identified end-user and walk every interactive
   path, not just that it renders. The widget lives in same-origin IFRAMES — host
   DOM / a11y-tree queries see nothing; read `iframe.contentDocument` or
   screenshot (ready-made snippets: [patterns.md](references/patterns.md) →
   Always verify → Driving it in a real browser). Don't claim "done" before this.

## Going deeper

[references/patterns.md](references/patterns.md) — the judgment the schema can't
carry: choosing the content type, segments as the targeting primitive, stable
selectors. Mechanics (fields, step types, constraints, what each type needs) come
from `get_authoring_guide`, not here.

Working request bodies: [references/examples.md](references/examples.md).

## Notes

- **Targets**: only `tooltip` steps (and launchers/element conditions) need a
  CSS-selector target. The runtime uses the FIRST match, so make the selector
  unique (a stable `id`/`data-*`) or pin it with `nth`; `text` only validates the
  chosen element's exact content — it can't find an element on its own. No stable
  selector? Add one in the app source (not a reused component), not a brittle
  `nth-child`. (Full rules: [Stable selectors](references/patterns.md).)
- **Navigation**: give a step a `key` and a button `{ "type": "goto_step",
  "step": "<key>" }` elsewhere in the same write resolves to it — author a whole
  flow in one call.
- **Strict by design**: the API fulfils your request exactly or errors — it never
  silently does something else. Read `validate` errors; they're precise.
