---
description: Build and publish a Usertour onboarding experience (flow / checklist / launcher / banner) from a description
argument-hint: [what to build]
allowed-tools: [Read, WebFetch]
---

# Build a Usertour onboarding experience

The user wants to build: $ARGUMENTS

Follow the `usertour-content-authoring` skill. Steps:

1. Read `skills/usertour-content-authoring/SKILL.md` and
   `skills/usertour-content-authoring/references/patterns.md`.
2. Call the `get_authoring_guide` MCP tool with no args (core conventions + a
   table of contents), then fetch the sections for the content types you'll
   build in one array call.
3. **Design first** — decide who / one action / when / done, then pick the
   content type(s) using the skill's design defaults (parallel tasks → checklist,
   not a grand tour; first-run flow 3–5 steps; trigger on context, not on load).
   See SKILL.md "Design before you author" and patterns.md.
4. `list_themes` and pick a `themeId`.
5. For each piece: `get_content_schema({ type })` → `create_content` →
   `update_content_version` → `validate_content_version` (fix errors) →
   `publish_content`.
6. Use real, stable selectors from the target app. If you don't know the app's
   DOM, ask the user for the page/URL and the elements to anchor to.
7. **Verify it renders** — publishing only *stores* the content; it shows only
   once the app loads the SDK and calls `identify()` with the SAME id you targeted
   (the `externalId`). Verify by loading the app as that identified user. If it
   doesn't appear because the SDK isn't installed/wired, that's the rendering half
   — hand off to the `usertour-sdk-install` skill (not this command's job).

Confirm the plan with the user before creating content if the request is
ambiguous. Never invent field shapes — fetch them from `get_content_schema`.
